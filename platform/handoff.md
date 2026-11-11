# Handoff: гостевая ссылка на встречу (FUSIO-1307)

Состояние на 2026-11-11 (вечер). Продолжать с этого места.

## 2026-11-11 (вечер): вкладка «Встречи» — клик открывает карточку встречи

Ветка `FUSIO-1307-livekit-shared-link`, стенд `dev/docker-compose.yaml`, фронт :8087, воркспейс `test`,
аккаунт `4acd5196-befc-4eb7-893b-815587ab84ca` (второй «Зайцев Андрей», у него 2 встречи; у `41633da8-…` — 14).

**Сделано и проверено на стенде.**

- Клик по встрече в календаре вкладки «Встречи» открывает карточку `EditScheduledMeeting`
  (кнопки `Edit`, `Join meeting`, чекбокс `Remind before`), а не календарный попап `EditEvent`.
  Проверено Playwright'ом: `hasEdit=true`, `join=true`, `remindBefore=true`, заголовка «Recurring event» нет.
- Правка названия в карточке сохраняется в **мастер** события: после `Save` и перезагрузки в БД
  `event.title = 'Meeting create 3643 edited'`. То есть 9.14 (правки не сохранялись) закрыт: раньше попап писал
  во вхождение серии.
- Новый модуль `plugins/calendar-resources/src/eventCard.ts` (+ `__tests__/eventCard.test.ts`, 5 кейсов) и его
  подключение в `EventElement.svelte` (3-дневный/недельный вид) и `Day.svelte` (месяц). Удалены промежуточные
  варианты: `openAsDocument.ts`, `cardMixin.ts`, `calendar.metadata.OpenAsDocument` и `setMetadata` в
  `love-resources/src/index.ts`.
- Пакет `calendar-resources`: 12/12 тестов, `pnpm run build` чистый; образ `pod-front` собран
  `pnpm docker:build --to @hcengineering/pod-front --force` (0 ошибок) и развёрнут, `nginx` перезапущен.

**Как это работает (чтобы не искать заново).** Календарь запрашивает `calendar.class.Event` без lookup, поэтому
у плитки нет поля `love:mixin:MeetingEventLink` — ни `hasMixin`, ни `mixinOrClass` (нужен прокси `$__mixin`) миксин
не видят. `eventCard(client, event, calendar.component.EditEvent)` поэтому ищет миксин сам: перебирает
`getAllPossibleMixins(event._class)`, оставляет те, у кого `view.mixin.ObjectEditor` — не календарный редактор,
и проверяет привязку через `client.findOne(mixin, { _id })` (миксин — документ с id расширяемого дока).
Для вхождения (`recurringEventId`) мастер читается **через миксин** (`{ eventId, access: Owner }`), чтобы в объекте
были и поля миксина; открывается `showPopup(editor, { object }, 'content')`.

**Грабли, которые стоили времени (все проверены на практике).**

- `setMetadata(calendar.metadata.OpenAsDocument, …)` из `love-resources` **не работает**: плагин грузится лениво,
  а `CalendarView.update` строит запрос на монтаже — метаданные приходят после.
- `hierarchy.as(doc, mixin)` + `openDoc` карточку **не дают**: `getObjectLinkFragment` кладёт в URI только
  `_class`/`_id`, панель перечитывает документ и снова теряет миксин (открывается «Recurring event»).
- `EditScheduledMeeting.openEditor` берёт `permanentMeeting` из live-запроса по `master._class` — документ без
  миксина, поэтому `settings` пуст и кнопка `Edit` уходит в календарный редактор (это и есть незакрытый 9.15).
- `pnpm docker:build` без `--force` отдаёт закэшированный бандл; браузер к тому же держит старый чанк — перед
  проверкой Playwright'ом снимать кэш (`Network.setCacheDisabled`) и сверять имя чанка с `dist` в контейнере.
- `services/love` jest резолвит `@hcengineering/love` из `lib/` — после правок сигнатур сначала `pnpm build --to`.

**Что осталось (по порядку).**

1. **9.15** — `EditScheduledMeeting.svelte`: читать `permanentMeeting`/`reminderBefore` явным запросом
   `client.findOne(love.mixin.MeetingEventLink, { _id })`, иначе `Edit` в карточке не ведёт в `PermanentMeetingForm`
   и Repeat (9.13) через карточку не проверить. Тест: у `test1`/`test2` `Edit` → `PermanentMeetingForm` → смена
   периода → `rules` у серии и `repeatEvery` у встречи. Для события без `permanentMeeting` фолбэк на календарный
   редактор правильный («Meeting create 3643 edited» — именно такой случай).
2. **9.16** — создание из вкладки: галочка «Create meeting» в `calendar.component.CreateEvent` по умолчанию выключена,
   поэтому создаётся событие без миксина (в календаре встреч не видно). Включить по умолчанию на этой вкладке или
   дать отдельную кнопку «Новая встреча» → `love.component.PermanentMeetingForm`.
3. **8.3** — скриншоты для `docs/images/` (нужен стенд документации; локально не снимались).
4. Документация: `docs/love.md` (строки списка: ближайшее вхождение, копирование ссылки, удаление/выход, группы)
   и `docs/features/love.md` — отметить, что клик по встрече в календаре открывает карточку.
5. «Покинуть» в списке постоянных встреч руками не проверено — нужен второй аккаунт (не создатель); в CI это 10.2.

## 2026-11-11 (проверка на стенде): модель починена, найдены три UI-дефекта списка

Проверено на стенде Playwright'ом (воркспейс `test`, фронт :8087).

**1. `ancestors not found: love:class:MeetingGroup` — исправлено.** `TMeetingGroup` не был перечислен
в `builder.createModel(...)`; после правки `pods/server: pnpm run get-model` отдаёт 4 записи
(`love:class:MeetingGroup`, `_name`, `_order`, `PermanentMeeting_group`). Ошибка в консоли больше
не воспроизводится. Сборка `model-love` 0 ошибок, jest `models/love` 9/9.

**2. Список «Постоянные встречи» был пуст при живых данных — две причины, обе исправлены.**

- Запросы списка создавались одноразовым `$:` сразу при монтировании. `Hall` монтируется ещё до
  подключения клиента (вкладка офиса рендерится на экране загрузки), а `LiveQuery`, созданный без
  `dontDestroy`, в `globalQueries` не попадает и после `setClient` не переподписывается — запрос
  оставался `pending` навсегда. Теперь запросы идут из `onClient(...)` из `@hcengineering/presentation`.
- Вторая и главная: `{#each sections() as section}` — Svelte 4 собирает зависимости **по тексту
  реактивного выражения**, а `sections()` читает `meetings`/`groups`/`series` внутри функции. Такой
  each не помечается грязным при приходе данных: в консоли `[PML] meetings 4`, в DOM — 0 строк.
  Теперь view-model считается в реактивном операторе `$: sections = buildSections(groups, meetings,
  series)`, где все три массива видны компилятору; строки несут `next`, `canCopyLink`, `canDelete`,
  `canLeave`.

**3. В списке не было удаления и выхода** (ТЗ п. 7–8: удаляет только создатель, остальные выходят).
Добавлены кнопки в строку: `Delete` (`isPermanentMeetingCreator` + `DeleteConfirmationPopup`,
живые сессии завершает серверный триггер) и `Leave` (`canLeaveMeeting`, `$pull` из `members`).

**Мелкие правки UI:** `Card` по умолчанию подписывает кнопку «Create» — в попапе ссылки стояла
неактивная «Create» (добавлены `canSave`/`okLabel`), в окне настроек теперь «Save» вместо «Create».

**Что подтверждено на стенде:** навигатор офиса (Встречи / Постоянные встречи / Этажи), окно
«Новая встреча» (название, дата/время, период, напоминание, пароль, участники), создание встречи →
попап «Guest link is ready» со ссылкой, список с 4 встречами, «Объединить в группу» → секция
«Команда» с двумя встречами (переживает перезагрузку — порядок и группы в БД), карточка встречи
с галкой «Постоянная встреча», которая открывает то же окно настроек, переименование через
настройки сразу видно в списке.

**4. Период («Repeat every») не переключался — исправлено.** После выбора `Day` лог показывал
`[REE] commit 1 day msec 86400000 known 604800000 hasOnChange true`, форма применяла значение
(`[PMF] changePeriod 86400000`), но следом приходило `[REE] sync from value 604800000 known 86400000` —
то есть родитель успевал отрендерить ребёнка со **старым** `period`, а `$:`-синхронизация по условию
`value !== known` сбрасывала единицу обратно в `Week`. Значение залипало навсегда.
Решение в `RepeatEveryEditor.svelte`: значение, которое редактор сам запросил, помечается `pending`,
и входящие пропы игнорируются, пока родитель не вернёт ровно это значение (`value === pending`);
`shown` хранит то, что реально показано, чтобы новые значения из документа всё же доходили.
Проверено: 8080 и 8087 — `Day → Month → Week`, число `3` даёт `3 Week`, сохранение и повторное
открытие настроек показывают записанный период (`1 Day` на 8080, `1 Month` на 8087).

**Осталось по спецификации:** п. 8.3 — скриншоты в `docs/images/`; обновить `docs/love.md`
(строки списка с ближайшим вхождением, копирование ссылки, удаление/выход, группы). «Покинуть» в списке
ручной проверкой не покрыто — нужен второй аккаунт (не создатель); в CI это задача 10.2 (стенд 8083).

## 2026-11-11: мелкие пункты — язык встречи, инвариант гостя, «мёртвые» поля

- **7.1 (язык)** — делать нечего: `CreatePermanentMeetingPopup.svelte` только хостит окно, язык берётся из
  интерфейса в `PermanentMeetingForm.save()` (`getCurrentLanguage()`) и уходит в `ScheduleContext.language`.
  Хардкода `'en'` в попапе и в `permanentMeetingSchedule.ts` нет.
- **7.2 (гость не получает `after`)** — инвариант держится двумя условиями `handleGuestInfo`: `isOver` не
  срабатывает для `kind: 'meeting'`, и ветка сессии проверяет `meetingDoc.meeting === undefined` (у сессии
  постоянной встречи `meeting` всегда задан). Закреплено тестом в `meetingLink.test.ts`: после завершённой сессии
  лобби отдаёт `live` (форма входа остаётся), `after` не отдаётся. `services/love` 232/232.
- **8.1–8.3 («мёртвые» поля)** — задачи устарели: `linkId` пишет сервис (`setMeetingLinkId`,
  `main.ts:548`), `meetingId` у `TMeetingSchedule` пишет booking-путь и читает `EditMeetingScheduleData.svelte`.
  BREAKING-изменения модели не нужно, `docs/features/love.md` править нечего.

Побочно: в тестовом фейке `resolveSession.test.ts` пришлось научить матчер понимать `$ne` и `$gte`/`$or` — после
`_class: { $ne: ReccuringInstance }` в `meetingMasterQuery` фейк перестал находить мастер.

## 2026-11-11: владелец постоянной встречи не выпадает из участников (этап 4)

Серверный инвариант уже стоял в `OnPermanentMeetingUpdate` (`server-plugins/love-resources/src/index.ts`):
недостающих владельцев триггер возвращает и в `owners`, и в `members` через `$push`, отдельно разобраны полная
перезапись и `$pull` (одиночный id, массив, `$in` — так снимает участника `SpaceMembers`), плюс защищён создатель
(`createdBy` → аккаунт). Триггер не может отклонить транзакцию, поэтому «правка без владельца отклоняется»
реализовано ответной правкой — ровно как согласовывали в `love-permanent-meetings` 6.2.
Проверка: `onPermanentMeetingUpdate.test.ts` — 15 тестов, зелёные.

Добавлено со стороны UI: `withOwners` (`plugins/love-resources/src/permanentMeetingSchedule.ts`), применён в
`PermanentMeetingForm.membersChanged` — после любой правки списка владельцы возвращаются, поэтому форма не
показывает ложную картину и не отправляет список без владельца. Проверка: 3 unit-теста (76/76 в пакете), плюс
живая проверка на 8087: снятие владельца в редакторе участников оставляет «2 members».

**Заведены три новых дефекта от владельца (вкладка «Встречи», задачи 9.13–9.15 в `love-permanent-meetings`):**
не меняется период повторения при редактировании, не сохраняются название и остальные параметры, не создаются
встречи. Все три надо воспроизводить под аккаунтом, у которого есть встречи (см. заметку про два аккаунта).

## 2026-11-11: миксин встречи живёт только на мастере серии (этап 3 `love-meetings-followups`)

Дыра: `ReccuringInstance` в календаре того, кто перенёс вхождение, сохраняет `access: 'owner'`, поэтому одной
проверки `access !== owner` в триггере не хватало. Запись миксина на вхождение давала серии вторую идентичность,
а поиск мастера по `{ eventId, access: Owner }` мог выбрать это вхождение и сделать его всей серией.

Код:
- `server-plugins/calendar-resources/src/index.ts` — `onEventMixin` (экспортирована) и `onEventUpdate` отсекают
  класс `ReccuringInstance`; в `onEventMixin` отброшенная запись пишется в лог
  (`Meeting identity mixin written to an occurrence, ignored`) с `eventId`/`objectId`/`modifiedBy`.
- `plugins/love/src/utils.ts` — `meetingMasterQuery` возвращает `_class: { $ne: calendar.class.ReccuringInstance }`.

Проверки: новый `server-plugins/calendar-resources/src/__tests__/eventMixin.test.ts` (3 кейса: копия не получает
миксин, вхождение не разгоняется по серии + пишется лог, мастер разгоняется), пакет 5/5; `meetingMasterQuery.test.ts`
обновлён и дополнен, `plugins/love` 50/50; образы `pod-server`/`pod-front`/`pod-love` пересобраны, `transactor`
пересоздан без ошибок триггеров.

**Важно про стенд (2026-11-11, вечер):** в воркспейсе `test` два аккаунта с одинаковым именем «Зайцев Андрей» —
`41633da8-…` (14 постоянных встреч) и `4acd5196-…` (2). Сессия 8087 сейчас под вторым, и список показывает только
его встречи: `PermanentMeeting` — это Space, видимость идёт по `owners`/`members`. Это не регрессия, но при ручных
проверках надо смотреть, под каким аккаунтом залогинен (смена аккаунта случилась через invite-вход на 8080).

## 2026-11-11: перенос вхождения серии учитывается (2.1-2.4 `love-meetings-pr428-gaps`)

Дефект: пользователь переносит вхождение в календаре, а ссылка открывает его по старому времени;
отменённое вхождение (`ReccuringInstance` с `isCancelled`) тоже продолжало открываться.

Код:
- `plugins/love/src/utils.ts` — новый тип `OccurrenceOverride`; `meetingOccurrences(master, from, to, overrides)`
  (override убирает своё вхождение, перенесённое добавляет новое время, отменённое не добавляет) и
  `resolveOccurrence(master, now, overrides)`.
- `services/love/src/workspaceClient.ts` — `occurrenceOverrides(master, now)` читает `calendar.class.ReccuringInstance`
  по `recurringEventId` системным клиентом, выборка `$or: [{date: {$gte}}, {originalStartTime: {$gte}}]`;
  используется в `findMeetingTarget` (для `nextOccurrence`/TTL ссылки) и в `resolveSession` (окно входа и `occurrence`).

Проверки: `plugins/love` 49/49 (4 новых кейса: отмена, перенос на часы, на день, отмена+перенос),
`services/love` 231/231 (отменённое вхождение даёт `no-occurrence` + следующее время, перенесённое открывается
только в новое время). Образ `pod-love` пересобран и накатан, гостевая ссылка после перезапуска сервиса
по-прежнему входит (`/guestJoin` 200, подключено). Sanity-сценарий переноса — задача 2.5, нужен стенд 8083.

**Грабли:** `services/love` в jest берёт `@hcengineering/love` из `lib/`, а не из `src` — после правки
сигнатуры в `plugins/love` нужно `pnpm build --to @hcengineering/love`, иначе старый код молча игнорирует
новый аргумент и падают тесты сервиса (не модели).

## 2026-11-11: сквозной прогон 8.1 и остаток 9.8 закрыты

Прошёл на 8087 (Playwright, один аккаунт), результаты по шагам:

| Шаг | Результат |
| --- | --- |
| Создать встречу с паролем | попап «Guest link is ready» со ссылкой `…/meetings/<shortId>` |
| Гостевая ссылка | `/guestInfo` → форма с подписанным полем пароля → `/guestJoin` 200 → LiveKit подключён |
| «Merge into group» | попап «New group» с именем и чекбоксами → группа с 2 встречами |
| Переименование в настройках | список обновился сразу (`E2E A 7522` → `E2E A renamed`) |
| DnD строк внутри группы | порядок `B, A`, после перезагрузки сохранился |
| Удаление строки | `Delete object` → `Yes` → строка исчезла, группа осталась |
| Этажи: сворачивание | `.hulyNavGroup-content`: 0 → 620 → 0 px |
| Этажи: создание | плюс в шапке `FLOORS` → «Floor / Add a floor» → второй этаж |
| Этажи: DnD | шапка `Main` перетащена на `Second` → порядок `Second, Main`, выжил перезагрузку |
| Live-точка | при живой сессии в комнате `.live` появляется на её строке в навигаторе |

Грабли прогона (пригодятся для 8.3 и 10.2): строки списка — `div[data-id="permanent-meeting-row"]` с
вложенными кнопками (копировать / присоединиться / удалить), клик по самой строке открывает окно настроек;
синтетический `dispatchEvent('click')` по вложенной кнопке удаления подтверждение не открывает — нужен
настоящий клик (`locator.click()`); DnD работает через HTML5 drag, поэтому перетаскивать надо элемент
`[draggable="true"]` (шапку группы/этажа), а не обёртку.

## 2026-11-11 (гостевой вход): три причины, все исправлены

Владелец: «нельзя подключиться гостю» и «не видно инпута пароля». Воспроизведено на 8087 в отдельном
контексте браузера. Симптом: форма заполнена, `Join meeting` → «Missing invite information», и в сети
**ни одного** запроса `/guestJoin`; `guestInfo` отвечал `{"passwordRequired":true,...}` без `meetingId`.

1. **`GuestJoinPopup.join` требовал `meetingId`** (`guestToken == null || meetingId == null`). Для ссылки
   за паролем `/guestInfo` намеренно не отдаёт ни `meetingId`, ни `mode`, ни название — «nothing leaks
   before the password is checked». `meetingId` в попапе нужен только как ключ `localStorage`, сервер
   сам разбирает токен. Guard сведён к `guestToken == null`.
2. **Постоянная встреча не была «всегда открытой».** `createPermanentMeeting` писал
   `meetingAccess: defaultMeetingAccess` (`start: 'members'`), а `resolveGuestLink` открывает сессию
   только при `start: 'link'` — гость обязан был ждать, пока участник запустит встречу. Теперь создание
   пишет `{ ...defaultMeetingAccess, start: 'link' }`, а upgrade-шаг `permanent-meeting-link-open`
   проставляет политику уже существующим встречам.
3. **Комнату LiveKit создавал только `/roomToken`**, то есть участник. Гость получал 404
   `Meeting room not found`. Добавлен `GuestManager.ensureRoom` — те же metadata/agents, что у `/roomToken`.

Заодно: лобби постоянной встречи без сессии отдаёт `mode: 'live'` вместо `'before'` (иначе гость видел
«ждём хоста» и не мог войти вообще); поле пароля получило подпись (`data-id="guest-password-row"`);
шапка лобби больше не печатает `- null`, когда `workspaceUrl` нет.

**Проверено на 8087:** ссылка с паролем → `/guestJoin` 200, `data-id="guest-connected"`, кнопка «Leave room»,
в комнате AI-агент; ссылка без пароля → форма входа без поля пароля. Тесты: `love-resources` 73/73,
`services/love` 229/229 (обновлён тест лобби: постоянная встреча без сессии = `mode: 'live'`,
добавлен тест на `meetingAccess.start === 'link'`).

**Важно про стенд:** LiveKit в `dev/docker-compose.yaml` отсутствует — он поднимается отдельно
(`dev/run_livekit.sh`, нативный `livekit-server`, порт 7880). Без него `/guestJoin` и `/roomToken` падают
с `ECONNREFUSED …:7880`, а комната не создаётся. На этой машине бинаря нет, для проверки поднят
временный контейнер: `docker run -d --name dev-livekit-tmp --network dev_default -p 7880:7880 -p 7881:7881
-p 7882:7882/udp -v <tmp>/livekit.yaml:/livekit.yaml livekit/livekit-server --config /livekit.yaml`
(`enable_loopback_candidate: true`, без redis и webhook). Удалить: `docker rm -f dev-livekit-tmp`.

**Грабли стенда (записать в память):**

- `nginx` не стартует без `mail_server`, `mail_client`, `mailpit` (`host not found in upstream
  "mail_server:8097"`) — весь стенд отдаёт пустой ответ; поднимать их первыми.
- `upgrade-workspace` принимает **UUID**, а не имя (`workspace test not found` на имени).
  `-f` обходит сверку версии (`ws-operations.ts:336`), но `model_tx` воркспейса при этом не меняется:
  на этом стенде модель клиенту и серверу приходит из бандла, `model_tx` хранит только 19 висящих
  `TxUpdateDoc` (мусор, `load-model` их пропускает).
- `common/scripts/version.txt` поднят до **0.8.16**: фронт, `pod-server` и `pod-workspace` пересобраны
  под эту версию, иначе воркспейс встаёт на «Preparing workspace for new version… 0.8.15 => 0.8.16».
- Навигатор офиса скрывается флагом `localStorage['love-visibleNav']` — пустая левая панель это
  свёрнутый навигатор, а не баг.

## 2026-11-11: причина `ancestors not found: love:class:MeetingGroup` найдена

Диагноз (не устаревший образ, а код): в `models/love/src/index.ts` класс `TMeetingGroup` объявлен
(`@Model(love.class.MeetingGroup, ...)`, строка 438), но **не перечислен в `builder.createModel(...)`**
(строки 500–516). Поэтому в `model.json` создаётся атрибут `love:class:PermanentMeeting_group`
(он ссылается на класс), а самой транзакции `TxCreateDoc` для класса `love:class:MeetingGroup` нет.

Проверка, на которой это видно (локально, HEAD `c80264e1c3`):

```bash
cd pods/server && pnpm run get-model
python - <<'EOF'
import json, io
d = json.load(io.open('bundle/model.json', encoding='utf-8'))
hits = [t for t in d if 'MeetingGroup' in json.dumps(t)]
print(len(hits), [t['objectId'] for t in hits])
EOF
# 1 ['love:class:PermanentMeeting_group']  ← класса нет, атрибут есть
```

До правки на стенде ошибка воспроизводится в `AggregationMiddleware.subscribe`
(`plugins/view-resources/src/middleware.ts:80` → `Hierarchy.getAncestors`).

**Правка:** добавить `TMeetingGroup` в список `builder.createModel(...)` в `models/love/src/index.ts`.
Дальше: `pnpm build --to @hcengineering/model-love`, `pnpm docker:build --to @hcengineering/pod-server
--to @hcengineering/pod-front --force`, `up -d front transactor --force-recreate --no-deps`, рестарт nginx.

Сверка ТЗ (pasted-версия) с кодом на эту дату — в `## 2026-11-11: сверка ТЗ` ниже (после правки модели).

## 2026-11-10: список этажей, отвязка встречи от комнаты, booking-путь

**Ветка и стенд.** `FUSIO-1307-livekit-shared-link`. Все поды стенда (`front`, `transactor`, `workspace`, `account`)
пересобраны из ветки с `--force`, версия модели везде `0.8.15` (`common/scripts/version.txt` возвращён к значению ветки).
Воркспейсы: `test` (старый, его когда-то подняли образы из develop) и `testlove` (создан тулом из ветки:
`create-workspace testlove huly:cb732a46-4251-47e4-a2d2-6146285d797d`, владелец добавлен `assign-workspace`).

**Корректировки владельца (2026-11-10) — все выполнены в коде:**

1. Сетку этажа оставляем, но она видна только после клика на этаж в списке.
2. Этажи — список: сворачивание/разворачивание, перестановка местами (DnD), при развороте видны комнаты
   и участники, оттуда начинаем встречу.
3. «Повторить логику отображения комнат как в сообщениях» — строки комнат: аватар владельца офиса, название,
   имена участников, счётчик и зелёная точка живого звонка.
4. «Группировка по этажам как в документах (книга → страницы)» — этаж = секция `NavGroup`, комнаты внутри неё.
5. Убрать привязку регулярной/постоянной встречи к комнате: встреча — отдельная сущность, комната больше не
   выбирается ни в окне постоянной встречи, ни в форме календарного события.
6. Рядом с заголовком «Этажи» — плюс создания этажа для роли `Owner` (и `Maintainer`), видимый постоянно,
   а не только по ховеру.
7. Выровнять отступы иконок и текста в списке (по замерам рендера: заголовок этажа 109px, иконка комнаты была
   115px при тексте 139px и почти без зазора; теперь иконка комнаты на 109px, зазор до текста 10px).
8. Починить падение `DocNotifyContextCardHeader`: `objectTitle` у контекста уведомления бывает `undefined`.

**Что сделано в коде:**

| Файл                                                                                               | Что                                                                                                                                                                                                                                        |
| -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `plugins/love/src/types.ts`, `models/love/src/index.ts`                                            | `Floor.order?: number` (ручной порядок этажей); `MeetingSchedule.meetingId?: Ref<PermanentMeeting>` — было обязательным `Ref<MeetingMinutes>` и никем не писалось                                                                          |
| `plugins/love-resources/src/floors.ts` (новый)                                                     | `reorderFloors` (через `moveItem`) и `persistFloorOrder` (пишет только изменённые этажи)                                                                                                                                                   |
| `plugins/love-resources/src/__tests__/floorOrder.test.ts` (новый)                                  | 5 кейсов на порядок этажей                                                                                                                                                                                                                 |
| `plugins/love-resources/src/components/FloorsNav.svelte` (новый)                                   | этажи-секции `NavGroup` (fold, выбранный этаж, DnD), комнаты `NavItem` с аватаром владельца, участниками, счётчиком и live-точкой; клик по этажу → `/love/<floorId>`, клик по комнате → документ комнаты (`selectedRoomPlace` + `openDoc`) |
| `plugins/love-resources/src/components/OfficeNavigator.svelte`                                     | вместо плоского списка — `<FloorsNav />`; плюс создания этажа в слоте `extra` (рендерится всегда), роли Owner/Maintainer, клик не сворачивает секцию                                                                                       |
| `plugins/love-resources/src/components/Hall.svelte`                                                | ветка `$officeView === 'floors'` → плейсхолдер `love.string.SelectFloor`; сетка — только детальный вид этажа                                                                                                                               |
| `plugins/love-resources/src/stores.ts`                                                             | `officeFloors` сортируются по `order` (без order — в конец, по имени); `officeView` получает состояние `floors`                                                                                                                            |
| `plugins/love-resources/src/permanentMeetingSchedule.ts`, `components/PermanentMeetingForm.svelte` | комната убрана: `ScheduleContext.language`, `PermanentMeetingInput.room` удалён, тип/запись/транскрипция/приватность — дефолты, язык из `getCurrentLanguage()` (`@hcengineering/theme`, не `platform`)                                     |
| `plugins/love-resources/src/utils.ts`                                                              | `createMeeting` ставит миксин без комнаты; `createMeetingSchedule` создаёт `PermanentMeeting` и пишет его в `MeetingSchedule.meetingId` (серия не создаётся — у `Schedule` нет `eventId`)                                                  |
| `plugins/love-resources/src/components/MeetingData.svelte`                                         | вместо выбора комнаты — подпись `MeetingRunsInServiceRoom` (ключ добавлен в 12 локалей)                                                                                                                                                    |
| `plugins/love-resources/src/components/EditMeetingScheduleData.svelte`                             | кнопка «Постоянная встреча» — переход в карточку созданной встречи                                                                                                                                                                         |
| `models/notification/src/migration.ts`                                                             | guard `hasLegacyCollaborators` (`hierarchy.hasClass`) для `accounts-to-social-ids-v2` и `migrate-collaborators-v2`: апгрейд воркспейса падал на `class not found: notification:mixin:Collaborators` и модель не доезжала                   |
| `plugins/notification-resources/src/components/DocNotifyContextCardHeader.svelte`                  | null-guard перед `objectTitle.startsWith('#')`                                                                                                                                                                                             |

**Проверки.** `model-love`, `love-resources`, `love-assets`, `model-notification`, `notification-resources` — 0 ошибок;
jest в `love-resources` — 16 suites / 72 теста. Образы `pod-front`, `pod-server`, `pod-workspace` собраны `--force`.

**Команды за сегодня (просьбы владельца и рабочие шаги):**

- «Сетку давай оставим, но она будет отображаться только при клике на этаж в списке».
- «Нужно чтобы мы могли отображать этажи в списке, менять их местами, сворачивать-разворачивать и при развороте
  там отображались все комнаты и участники, с чем мы можем начать встречу».
- «Отвязываем и не завязываемся от комнаты, это по сути отдельная сущность будет».
- «Рядом с кнопкой Этажи надо добавить плюсик для роли владельца» + «посмотри какой отступ у иконок и текста
  и проставь отступы».
- «Как 1 и 2 пункт выполнишь — делай дальше по плану» → взят booking-путь (задачи 2.3–2.4).
- «Давай миграцию» → починен апгрейд воркспейса (guard в `models/notification`).
- Рабочие команды стенда: `pnpm docker:build --to @hcengineering/pod-front --to @hcengineering/pod-server --force`,
  `up -d front transactor --force-recreate --no-deps`, `docker compose -f dev/docker-compose.yaml restart nginx`
  (обязательно: nginx кэширует DNS upstream, иначе 502), `pnpm -w bundle --to @hcengineering/tool` +
  `node ./bundle/bundle.js upgrade-workspace <ws> -f` (окружение — см. `dev/tool/package.json`, плюс
  `ACCOUNT_DB_NS=global_account`).

**Осталось. НАЧАТЬ С ЭТОГО:**

> **Ошибка, с которой начинать выполнение:**
> `ancestors not found: love:class:MeetingGroup`
> (`Hierarchy.getAncestors` → `getAllAttributes` → `AggregationMiddleware.subscribe` в `plugins/view-resources/src/middleware.ts:80`).
> Значит в модели воркспейса нет класса `love:class:MeetingGroup`, хотя он объявлен в `models/love/src/index.ts`
> (коммит `9329c65301`). Где копать:
>
> 1. `model_tx` воркспейса (`testlove` = `cae72b89-e884-474e-b736-36df073a5bc3`): там 19 висящих `TxUpdateDoc`,
>    класс `love:class:MeetingGroup` среди них не встречается — проверить, применяются ли модельные транзакции
>    при открытии воркспейса и кто их применяет (`transactor` vs `workspace`, `WS_OPERATION=all+backup`).
> 2. Сверить, что `models/love` в образе содержит класс: `TMeetingGroup` в `createModel`-списке, `lib/model.json`
>    в `pod-server`/`pod-workspace` — и что образы собраны из одной и той же ревизии.
> 3. После применения модели: раздел «Постоянные встречи», создание встречи, `EditPermanentMeeting` и группы
>    (`MeetingGroup`) должны открываться без ошибок.
> 4. Апгрейд воркспейса теперь проходит (`skip accounts migration: no legacy mixin in the model`) — проверить, что
>    он не «проглатывает» шаг применения модели (у `upgrade-workspace -f` в логе: `load-model`, `pre-migrate`,
>    `migrate`, `build local model`, `upgrade-workspace done`).

Прочее из остатка:

- Ручная проверка booking-пути на стенде: «Публичное расписание» → сохранение создаёт `PermanentMeeting`,
  в панели видна кнопка «Постоянная встреча».
- Ручные проверки из `love-recurring-meetings`: 1.4, 2.1–2.2, 4.1, 4.5, 5.4.
- Открытые решения: 5.3 (кто хозяин участников серии и постоянной встречи), 3.5 (`repeatEvery` отдельным
  атрибутом в панели), 2.3–2.4 (booking — код сделан, проверки за владельцем).
- Документация `docs/love.md` и `docs/features/office-meetings.md` под новые правила (список этажей, встреча без
  комнаты) не обновлялась — по решению владельца доки пока не трогаем.

## 2026-11-06 (вечер): новый объём — «Постоянные встречи» как полноценная встреча

- Владелец переформулировал раздел «Постоянные встречи» (9 пунктов: одно окно создания/настройки с датой/временем,
  регулярностью, участниками, напоминанием, паролем и комнатой; гостевая ссылка; таблица с действиями; карточка
  с отметкой; вкладка «Встречи»; режимы правки «текущая/все будущие»; права удалить/покинуть; группы с DnD).
- Заведено изменение **`openspec/changes/love-permanent-meetings/`** (валидно, 31 задача, 3 capability:
  `love-permanent-meetings`, `love-permanent-meetings-groups`, `love-meeting-permissions`).
- Ключевое решение (design.md, решение 1): модалка создаёт **серию события + `PermanentMeeting`**, связанную через
  `MeetingEventLink.permanentMeeting` — расписание, вхождения и сессии остаются календарными, а не второй
  реализацией на `PermanentMeeting`.
- **Реализация не начата:** ждём подтверждения открытых вопросов из `design.md` (обязательность расписания,
  плоские группы, «создатель» для старых данных, пароль только на установку).

## 2026-11-06 (день): дата/время в форме и удаление постоянной встречи

- **Дата/время применялись со второго раза.** Причина в календаре: `EventTimeEditor.svelte` получал от `DateEditor`
  событие `update` с новым значением, но игнорировал `ev.detail` и пересчитывал по устаревшему `startDate`. Теперь
  `dateChange`/`dueChange` принимают `CustomEvent<number | Date>` и ставят значение из `detail`.
- **Удаление постоянной встречи.** В модели зарегистрирован override `view.action.Delete` для `PermanentMeeting`
  (`visibilityTester: view.function.CanDeleteSpace`, режимы `context`+`browser`); для существующих воркспейсов —
  шаг миграции `permanent-meeting-delete-action` в `models/love/src/migration.ts`. Живые сессии завершает триггер.
  Дата/время у постоянной встречи не бывает (она открыта всегда); редактируются имя и период `repeatEvery` в панели.
- Образы пересобраны: `front:latest` и `transactor:latest` (свежие, с `--force`). После рестарта контейнеров
  `front` и `transactor` на стенде заработают: форма даты/времени, кнопка удаления во вкладке «Постоянные встречи».
- Спека: `love-recurring-meetings`, добавлены 6.8 и 6.9.

## 2026-11-06 (ночь): ревизия кода, документация

- **Сборка падала на Windows.** `bundle:template` в `services/love/package.json` снова копировал `index.html`
  через `fs.cpSync` — на Windows тот отдаёт `errno 0` / «The operation completed successfully» на `unlink`
  занятого файла, и это валило всю цепочку `pnpm docker`. Подключён лежавший в репозитории помощник
  `common/scripts/copy-file.js` (повторная копия вместо unlink). Проверено: `bundle:template` дважды подряд,
  затем `docker:build --to @hcengineering/pod-love`, `pod-server`, `pod-front` — все три образа собрались.
- **Про кэш:** если после правок клиента в логе фазы стоит `(cached)`, собирайте с `--force` — при ручной
  проверке `pod-front` без `--force` весь бандл взялся из кэша; с `--force` пересобралось 270 пакетов и образ.

- **Ревизия клиентского кода** (не тестов) по двум коммитам — найдено и исправлено четыре артефакта:
    1. вкладка «Встречи» показывала каждое перенесённое/отменённое вхождение отдельной встречей (календарь зеркалит
       миксин на все копии с тем же `eventId`); отсечено `baseQuery={{ recurringEventId: { $exists: false } }}`
       в `Hall.svelte`. По `originalStartTime` фильтровать нельзя — он есть и у мастера серии;
    2. напоминания: `setInterval` не снимался при повторном `onClient` (после релогина в той же вкладке напоминания
       дублировались), а запрос тянул все события без нижней границы; теперь один таймер с обновляемой ссылкой
       на клиент и запрос двумя ветками (серия — по `rules`, событие — по окну `date`);
    3. `RepeatEveryEditor` не сохранял значение в панели свойств: он зарегистрирован атрибутным редактором, а такому
       `onChange` не передаётся; теперь без `onChange` пишет сам через `client.diffUpdate`;
    4. `MeetingData` не инициализировал `isMeeting` из `$state`.
- Известное и не наше: `TMeetingSchedule.meetingId!` объявлен обязательным и никем не пишется (закрывается
  бэклог-задачей о booking-пути); в `createMeeting` мёртвый блок `navigateUrl` из апстримного Huly PR #6212.
- **Документация обновлена:** `docs/love.md` — §2.5 (поля `reminderBefore`/`permanentMeeting`, отмена и удаление,
  правило «триггер без Trigger-документа не работает»), §2.6 (`repeatEvery`), новые §7a.5 «Напоминания о вхождениях»
  и §7a.6 «Встреча из планировщика»; `docs/features/office-meetings.md` — путь создания и строка пользовательской
  документации.
- Спека: `love-recurring-meetings` **17/28**, валидна; `openspec validate --all --strict` — 4 passed.
- Сборки: `@hcengineering/love-resources`, `@hcengineering/model-love`, `@hcengineering/model-server-love`,
  `@hcengineering/server-love-resources` — 0 ошибок; тесты серверного пакета 53/53.
- **Осталось (за вами):** пересобрать образы `front`, `love` и `transactor` (`pod-server`) и пересоздать контейнеры;
  ручные проверки 1.4, 4.1, 4.5, а также 5.4 (сдвиг периода у идущей серии).

## 2026-11-06 (вечер): группа 4 сделана в коде, найден мёртвый триггер

- **Баг найден и исправлен:** `OnPermanentMeetingUpdate` (защита владельца постоянной встречи, коммит
  `a579404650`) никогда не вызывался — функция и ключ в карте триггеров были, а ресурс `serverLove.trigger` и
  документ `serverCore.class.Trigger` не заводились. Добавлено: ресурс в `server-plugins/love/src/index.ts`,
  Trigger-документ в `models/server-love/src/index.ts`, миграция `serverLoveOperation`
  (`models/server-love/src/migration.ts`, состояние `add-permanent-meeting-trigger`), регистрация операции
  в `models/all/src/migration.ts`. Правило: триггер без Trigger-документа не работает и в старых воркспейсах
  сам не появится — нужна миграция (как у бейджа карточки).
- **Отмена одного вхождения завершает его сессию.** Календарь отменяет вхождение заменой на копию
  с `isCancelled: true` (`deleteRecHandler`, `plugins/calendar-resources/src/index.ts:87`), а не через `exdate`.
  Поэтому в `OnEventUpdate` добавлены две ветки: `TxCreateDoc` с `isCancelled` в атрибутах и `TxUpdateDoc`
  с `isCancelled`; сессия ищется по `eventId` + `occurrence` (`originalStartTime`), остальные вхождения не трогаются.
- **Удаление постоянной встречи завершает её живые сессии** (`TxRemoveDoc` в `OnPermanentMeetingUpdate`,
  запрос по `meeting`). Общий помощник `finishLiveSessions(control, query)` заменил три копии одного и того же цикла
  в `server-plugins/love-resources/src/index.ts`.
- Тесты: `server-plugins/love-resources` — 53/53 (было 46; +4 кейса отмены вхождения, +3 кейса удаления постоянной
  встречи). Сборки `@hcengineering/model-server-love`, `@hcengineering/server-love-resources` — 0 ошибок.
- Спека: `love-recurring-meetings` 12/24, валидна. Закрыты 4.2–4.4; открыты ручные проверки (4.1, 4.5),
  документация (5.5) и пересборка стенда (5.6) — **образ `transactor` нужно пересобрать: триггеры живут в нём**.

## 2026-11-06: спека приведена к коду, остаток по регулярным созвонам — в одном изменении

- Ветка `FUSIO-1307-livekit-shared-link`, HEAD `e2da22dca2`, рабочее дерево чистое кроме `.vscode/settings.json`.
  Ветка запушена. За два дня сверху PR #428 (`ca12e587e5`) легли два коммита владельца:
    - `a579404650` «Добавление экранов + создание постоянных встреч» — Storybook-экраны фичи, `copy-file.js` и фикс
      генератора на Windows, `PermanentMeetingMembersEditor`, `OnPermanentMeetingUpdate` (+9 тестов), `setMeetingLinkId`
      (+4 теста), режим `/guestInfo` при завершённой сессии постоянной встречи, ассеты `Knock`/`StopRecord`,
      правки `docs/love.md` и `docs/features/office-meetings.md`;
    - `e2da22dca2` «Реализация отображения встреч» — модель (`MeetingEventLink.reminderBefore` и `permanentMeeting`,
      `TPermanentMeeting.repeatEvery`), миграция `addMeetingEventCardBadge`, `period.ts` + тесты, `RepeatEveryEditor`,
      `MeetingEventBadge` и точка расширения карточки события в `calendar-resources`, клиентские напоминания
      `meetingReminders.ts`, свои шапки `EditScheduledMeeting`/`EditPermanentMeeting`, «сделать постоянной» в
      `MeetingData` + создание `PermanentMeeting` в `createMeeting`, вкладка «Постоянные встречи» в `Hall.svelte`,
      локализация 12 языков, `OnEventUpdate` (завершение сессий при удалении серии, добавление участников в сессию),
      тест `onEventUpdate.test.ts`.
- **Спека выровнена с кодом.** Заведено единое `openspec/changes/love-recurring-meetings/` (proposal, design со
  состоянием кода на 06.11 и открытыми вопросами, tasks — 5 групп по списку владельца, спеки трёх capability).
  10 из 23 задач закрыто кодом. `openspec validate --all --strict`: 4 passed. Группа 12 из `love-meetings-pr428-gaps`
  переехала туда же, в самом `pr428-gaps` осталась только справка (28/39). Черновик `love-planner-permanent-meeting`
  поглощён и перенесён в `openspec/changes/archive/`.
- **Что разошлось с прежним планом** (и переписано в design.md): связь «серия → постоянная встреча» живёт в
  `MeetingEventLink.permanentMeeting`, а не в `TMeetingSchedule.meetingId`; `MeetingSchedule.meetingId` теперь нужен
  только для пути планировщика и пишется фазой `post`; вкладка «Встречи» не смешивает типы — постоянные встречи
  вынесены в отдельную вкладку `officeView === 'permanent'`; миксин `MeetingEventLink` объявлен на
  `calendar.class.Event`, поэтому встреча из планировщика должна создаваться как серия событий.
- **Осталось по коду** (подробно — `love-recurring-meetings/tasks.md`): путь `calendar.class.Schedule`
  (`MeetingScheduleData`, `createMeetingSchedule`) не умеет ни серию, ни постоянную встречу — это группы 2.1–2.4;
  ручные проверки правки/удаления во вкладке встреч — 1.4, 4.1, 4.3, 4.4; обратная синхронизация участников — 5.3;
  решение по показу `repeatEvery` в панели — 3.5; документация — 5.5; пересборка и стенд — 5.6.
- **Открытые вопросы, требующие ответа владельца:** что именно считается «добавлением встречи в планировщике»
  (событие через `MeetingData` или запись расписания `Schedule`); что делать с уже открытой сессией при отмене одного
  вхождения; кто хозяин списка участников у серии, породившей постоянную встречу.

## 2026-11-05: остаток по регулярным созвонам заведён отдельным изменением

> **Устарело 06.11:** каталог `love-meetings-pr428-gaps` в этой копии есть (28/39 задач), а `love-recurring-meetings`
> заведено заново и валидно (10/23). Читать как историю.

- **Расхождение в копии:** `openspec/changes/love-meetings-pr428-gaps/` в этом репозитории **отсутствует** —
  `openspec list` показывает только `love-meeting-link` и `love-meetings-followups`, а в git-истории
  (`git log --all -- '*pr428-gaps*'`) этого каталога нет. Ссылки на него ниже и в `plan.md` описывают работу
  из другой копии; при продолжении их надо либо восстановить, либо переписать на живые изменения.
- **Заведено `openspec/changes/love-recurring-meetings/`** (валидно, 4/4 артефакта, 22 задачи) — остаток
  по регулярным созвонам пятью группами ровно по списку владельца:
    1. отображение в планировании на календаре;
    2. создание регулярной встречи из планирования с настройкой регулярности;
    3. доработка настроек регулярных встреч;
    4. проверка редактирования/удаления во вкладке регулярных встреч;
    5. проверка изменения регулярных встреч и их настроек.
- Там же `design.md` с фактическим состоянием кода на 05.11: планировщик (`PlanningCalendar.svelte`) не
  знает о встречах, создание идёт через `calendar.component.CreateEvent` + `DocCreateExtension` love,
  правило повторения покрывает только `freq/interval/count/endDate` из модели `RecurringRule`, вкладка
  «Встречи» = `love.viewlet.CalendarScheduledMeetings` на миксине. Открытые решения: что считать «задачей»
  в планировании, что делать с живой сессией при удалении серии, что делать со ссылкой при смене настроек.
- В `proposal.md` того же изменения зафиксировано, что перенос вхождения (override) и внеплановый старт
  серии остаются вне объёма.

## 2026-11-03: спека приведена к коду, начат остаток работ

- Ветка `FUSIO-1307-livekit-shared-link`: `24eb5185cb` (наши правки) + `ca12e587e5` (PR #428), запушена; рабочее дерево чистое, кроме `.vscode/settings.json`.
- Заведено изменение `openspec/changes/love-meetings-pr428-gaps/` — единый план остатка (6 capability, 10 групп задач). `openspec validate --all --strict`: 3 changes passed.
- Документация приведена к коду PR: `docs/love.md` §8.3 вернул контракт `/guestJoin` (потерян при слиянии), `docs/features/office-meetings.md` переписан с `startOnLink`/`Scheduled` на миксин-модель, причина `stale` в таблице пользовательской документации поправлена.
- Устаревание помечено: `plan.md`, `diagrams.md`, `love-meeting-link` (proposal/spec/tasks), `love-meetings-followups/tasks.md` — статусы по `audit-pr-428.md` §2.
- Код за 03.11 (не закоммичен):
    - `services/love/src/guests.ts` — `/guestInfo` не отдаёт `after` завершённой сессии постоянной встречи (`meetingDoc.meeting !== undefined`), иначе лобби врало про «встреча кончилась».
    - `server-plugins/love-resources/src/index.ts` — новый триггер `OnPermanentMeetingUpdate`: правка `members` без владельца исправляется обратно (`$push` отсутствующих `owners`). Триггер не может отклонить транзакцию, поэтому инвариант держится ответной правкой; транзакция исправления несёт `$push`, поэтому зацикливания нет.
- Открыто из начатого: UI-запрет убирать владельца (задача 3.2) — участников постоянной встречи правит общий `SpaceMembersEditor` из `contact-resources`, он не знает про `owners`; нужно решение: передавать `object` в презентер атрибута и завести свой редактор для `love.class.PermanentMeeting` или править общий.
- Следующий шаг по плану: группа 2 «Перенос вхождения» отложена на следующую итерацию (решение 2026-11-03, источник override не согласован). Дальше по плану — группа 5 (навигатор, ручные проверки) и группа 6 (мёртвые поля модели, **BREAKING**).
- Закрыто 03.11 после уточнений: группа 3 целиком — серверный триггер + свой редактор участников `PermanentMeetingMembersEditor` (`createAttributePresenter` на `love.class.PermanentMeeting, 'members', 'array'`), общий `SpaceMembersEditor` не тронут; этап 5 «внеплановый вход по ссылке» — **не реализуем**, сценарий закрывает `PermanentMeeting` (`design.md` Decisions 7, отметка в `plan.md` §5).
- 03.11, после сверки с PR #428: модель и типы возвращены к паритету с PR (поля `linkId` и `TMeetingSchedule.meetingId` на месте, контроль `git diff 46537cd2c8`), а расхождение «объявлено — не пишется» закрыто записью shortId: `WorkspaceClient.setMeetingLinkId` + вызов из `GET /meetingLink` (4 теста, `services/love` 236/236).
- Запрошено в Storybook (задача 11 в `tasks.md`): экраны фичи рисуются кодом в `dev/design/screens/love-meeting-link/`, регистрация в `dev/design/design.json` + `generate.mjs`/`generate-screens`, стиль — как в `dev/design/screens/fusio-1613/`; скриншоты для docs — со стенда после деплоя PR.
- Storybook сделано 03.11: `dev/design/screens/love-meeting-link/` (`_meeting.js`, `01-meeting-link-popup.js`, `02-permanent-meeting.js`, `03-guest-lobby.js`), секция `Meetings` в `dev/design/design.json`, истории `dev/storybook/src/design/generated/stories/meetings-page-1..3.stories.ts`; `pnpm run build-storybook` зелёный. Попутно починен `dev/storybook/scripts/generate-screens.mjs` (на Windows падал из-за абсолютного пути в `await import` — теперь `pathToFileURL`). Подпись: секция НЕ должна называться `Screens` — slug совпадает с `Intabia Screens` и истории перезаписывают друг друга.
- 03.11, сборка фронта: `@hcengineering/love-assets` не собирался — при cherry-pick файл остался в develop-версии (спрайт `@hcengineering/ui/assets/icons.svg`) без ассетов `Knock`/`StopRecord`, которые PR добавил в `love.icon`. Добавлены (`#door-knock`, `#square-fill`).
- **03.11, откат рефакторингов в файлах слияния (решение владельца).** `authorizeTarget` вернулся в `main.ts` (`linkAuth.ts` и 7 его тестов удалены), фильтр организатора вернулся в `EventRsvp.svelte` (`rsvpInvitedCount` и 3 теста удалены), ссылки на картинки и формулировка `meetingAccess.start` в `docs/love.md` — как в слиянии; строка `linkId` — авторский текст (поле теперь пишется при выдаче ссылки). Правило записано в `design.md` Decisions 8: файлы, разрешённые при слиянии, не переписываем, правки — только аддитивные и после согласования. Тесты после отката: `services/love` 229, `calendar` 80, `calendar-resources` 7 — зелёные.
- 03.11, стенд: образы `love` и `front` пересобраны с `--force` (238 с) и контейнеры пересозданы. При пересоздании всплыло: `dev-nginx-1` ушёл в цикл рестартов — `host not found in upstream "mail_server:8097"`, то есть без сервисов `mail_server`/`mail_client` nginx не поднимается вовсе. Лечится `docker compose -f dev/docker-compose.yaml up -d mail_server mail_client mailpit` и рестартом nginx; после этого http://localhost:8087 отвечает 200.
- 03.11, стенд (продолжение): триггеры живут не в `pod-love`, а в `pod-server` (контейнер `transactor`) — образ собран с `--force` (361 пакет, 354 с), `transactor` пересоздан, `OnPermanentMeetingUpdate` есть в `bundle.js`. По пути починен тип в `onPermanentMeetingUpdate.test.ts` (`$push.members` — это `Position<T> | T`, прямое `.$each` не типизируется). На стенде сейчас новый код везде: `front`, `love`, `transactor`.
- **Sanity на этом стенде не гоняется.** `tests/sanity` нацелен на отдельный стенд `tests/docker-compose.yaml` (проект `sanity`, порт 8083, воркспейс `sanity-ws`), а dev-стенд живёт на 8087 без `sanity-ws`. Подготовка: `cd tests && ./do-test.sh` (нужны образы всех подов: `pnpm docker`), либо `prepare-pg.sh` без сборки, если образы актуальны. Памяти на машине ~3 ГБ свободно при 38 сервисах — риск OOM.
- Тесты 03.11: `server-plugins/love-resources/src/__tests__/onPermanentMeetingUpdate.test.ts` (6/6) и два кейса режима в `services/love/src/__tests__/guests.test.ts` (18/18 в файле).
- Дальше 03.11: группа 6 закрыта (удалены `linkId` и `TMeetingSchedule.meetingId`), тест RSVP (`rsvpInvitedCount` в `plugins/calendar/src/utils.ts` + 3 кейса), гейт авторизации ссылки вынесен в `services/love/src/linkAuth.ts` и покрыт 7 кейсами (`linkAuth.test.ts`), документация `docs/love.md` очищена от несуществующих картинок и неверного описания `meetingAccess.start`. Прогоны: `services/love` 232, `plugins/calendar` 83, `calendar-resources` 7, `plugins/love` 45, `models/love` 9 — все зелёные; сборки `pod-love`, `love-resources`, `model-love` — 0 ошибок.
- Решение по владельцу воркспейса (2026-11-03): особых прав на ссылку у него нет и не будет — ссылку выдаёт участник, остальные подписываются (`authorizeTarget` уже так работает). Записано в `specs/guest-access` и в `proposal.md`.

> **2026-11-02: PR #428 применён на эту ветку.** Коммит `46537cd2c8` (`origin/FUSIO-1307`) накатан через
> `git cherry-pick -n` — **без коммита**, изменения лежат в индексе/рабочем дереве ветки
> `FUSIO-1307-livekit-shared-link`. Разбор и остаток работ — [`audit-pr-428.md`](audit-pr-428.md),
> отчёт о применении — §12 там же. Прежнее описание `startOnLink` ниже **устарело**: PR заменил эту механику
> на `meetingAccess.start` (`members` | `link`), статуса `MeetingStatus.Scheduled` больше нет, перезапуска
> завершённой встречи нет — сессию создаёт `resolveSession` сервиса love.

## Состояние после применения PR #428

- Ветка: `FUSIO-1307-livekit-shared-link`, база develop до 2026-10-30, сверху 108 файлов PR.
- Коммита нет: `git status` показывает ~111 изменённых/добавленных файлов (индекс + рабочее дерево).
- Сборка зелёная: `@hcengineering/love`, `model-love`, `love-resources`, `calendar-resources`, `pod-love`,
  `server-love-resources` — 0 ошибок.
- Тесты: `services/love` 223, `plugins/love` 45, `models/love` 9, `server-plugins/love-resources` 34,
  `plugins/love-resources` 32, `plugins/calendar` 80, `plugins/calendar-resources` 7 — все зелёные.
- Наши доработки поверх PR перечислены в `audit-pr-428.md` §12 (язык постоянной встречи, сегмент навигатора,
  гостевые имена и стабильная персона, RSVP `pending`, флаг пароля).
- Не сделано и требует решения: перенос вхождения (override), «владелец всегда участник» для `PermanentMeeting`,
  `linkId`, мелочи UI (`getEventMeetingLink` глотает ошибки, роль-гарды).

## Где работаем (историческая часть, до применения PR)

> Всё ниже описывает состояние **до** наката PR #428 и в коде уже неактуально: `startOnLink`, `MeetingStatus.Scheduled`,
> `restartMeeting` и `findJoinableScheduled` удалены слиянием. Читать как историю решения, а не как текущее состояние.

- Ветка: `FUSIO-1307-livekit-shared-link`, HEAD = `df6deb09e6` (это тег `v0.8.63` = `origin/develop`). Ветка создана от develop, **коммитов нет** — все правки лежат рабочим деревом.
- Каталог: `c:\Users\Андрей\Desktop\project\platform\platform`.
- Спецификация (OpenSpec): `C:\Users\Андрей\Desktop\project\ProjectSpec\platform`
    - `plan.md` — план этапов + отчёт по этой ветке + блоки «Деплой на dev-стенд».
    - `diagrams.md` — 11 Mermaid-схем, §10 гостевая ссылка, §11 последовательность входа гостя.
    - `openspec/changes/love-meeting-link/` — `proposal.md`, `specs/love-meeting-link/spec.md`, `tasks.md`. Валидно: `openspec validate love-meeting-link --strict`.
    - `openspec/changes/love-meetings-followups/` — спека для _другой_ ветки (`FUSIO-1307` с моделью F1/F4), здесь не применима.

## Что сделано (рабочее дерево, не закоммичено) — до PR #428

> **Устарело 2026-11-02:** таблица ниже — прежняя механика гостевой ссылки. После слияния от неё остались
> только две вещи, перенесённые на модель PR: валидация имени гостя (`400`) и стабильный `guest-link:<sha256>`;
> список актуальных правок — в начале файла и в `audit-pr-428.md` §12.

Суть: ссылка на встречу работает всегда — гость входит вне окна расписания и после завершения, встреча перезапускается, комната пересоздаётся с тем же `meetingId` и тот же short id.

| Файл                                                                          | Правка                                                                                                                                                                                                                                                                              |
| ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `models/love/src/index.ts`                                                    | `@Prop(TypeBoolean(), love.string.StartOnLink) startOnLink?: boolean` в `TMeetingMinutes`                                                                                                                                                                                           |
| `plugins/love/src/types.ts`                                                   | `MeetingMinutes.startOnLink?: boolean`                                                                                                                                                                                                                                              |
| `plugins/love/src/plugin.ts` + `plugins/love-assets/lang/*.json` (12 локалей) | ключ `StartOnLink` («Начинать по ссылке может любой»)                                                                                                                                                                                                                               |
| `services/love/src/workspaceClient.ts`                                        | `restartMeeting`: `status: Active`, `meetingEnd: undefined`, сброс `transcriptionState`/`recordingState`, ранний выход при `Active`; `activateMeeting` не тронут (защита от stale-клиента)                                                                                          |
| `services/love/src/guests.ts`                                                 | `/guestJoin`: перезапуск только при `startOnLink === true` (иначе прежние 403); создание LiveKit-комнаты, если её нет; `/guestInfo` отдаёт `canStart`; валидация имени (`400` без имени) и вывод `personToken` (`guest-link:<sha256(meetingId : имя)>`), если клиент его не прислал |
| `services/love/src/main.ts`                                                   | `/getToken`: владелец встречи (`owners`) или владелец workspace (`AccountRole.Owner`) переводит `Scheduled` → `Active` вне окна; роль запрашивается только когда это меняет решение                                                                                                 |
| `services/love/src/__tests__/guests.test.ts`                                  | 4 кейса флага + создание комнаты + «без имени → 400» + «без personToken → стабильный выведенный id»                                                                                                                                                                                 |
| `plugins/love-resources/src/components/EditMeetingMinutes.svelte`             | кнопка-переключатель `startOnLink` в шапке панели встречи (только владелец), `data-id="meeting-toggle-start-on-link"`                                                                                                                                                               |
| `plugins/love-resources/src/components/guest/GuestMeetingApp.svelte`          | лобби без тупиков в `Scheduled`/`Finished`; при `canStart === true` не срабатывает `GUEST_ONLY_TIMEOUT_MS` (30 с) — иначе гость выкидывался из пустой комнаты                                                                                                                       |
| `plugins/love-resources/src/meetings.ts`                                      | `findJoinableScheduled` берёт `Scheduled`-встречу также при `startOnLink` или для владельца встречи/workspace                                                                                                                                                                       |
| `plugins/love-resources/src/components/Floor.svelte`                          | кнопка «Добавить комнату» для `AccountRole.User` (обработчик `openAddRoom`); «Редактировать офис» остаётся за `Maintainer`                                                                                                                                                          |
| `docs/love.md` §8.3, `docs/features/office-meetings.md`                       | контракты `/guestJoin` и `/getToken`, кнопка создания комнаты; строка пользовательской документации `communication/invite-link` → `stale`                                                                                                                                           |

Мёртвый `plugins/love-resources/src/components/EditMeetingData.svelte` **не** используется: он только экспортируется, нигде не рендерится. Правки из него откачены.

## Что задеплоено на стенд (сейчас запущено)

- `dev-front-1` — образ пересобран (199 с) после фикса `Floor.svelte`; в `dist/2358.13e868757aacb0a8.js` есть `meeting-toggle-start-on-link`, `AddRoom`, `keeping the guest alone`.
- `dev-love-1` — образ `intabiafusion/love:latest` пересобран, контейнер пересоздан.
- `livekit-server` — **вне compose**, отдельный контейнер (`livekit/livekit-server:latest`, конфиг `C:\Users\Андрей\livekit-dev.yaml`, webhook `http://host.docker.internal:8087/_love/webhook`, ключи `devkey/secret`, `whkey/whsecret`, Redis стенда, порты 7880/7881/7882). При запуске `dev/run_livekit.sh` его остановить, иначе конфликт портов.
- `dev-love-agent-1`, `dev-livekit-egress-1`, `dev-postgres-1`, `dev-minio-1` — как обычно.
- Фронт: http://localhost:8087, сервис love: порт 8096.

## Проверено (повторяемо)

| Проверка                                                  | Результат                                                                                        |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `openspec validate love-meeting-link --strict`            | валидно                                                                                          |
| `get_errors` по всем правленым файлам                     | чисто                                                                                            |
| `pnpm build --to @hcengineering/pod-love`                 | 47/47, 0 ошибок                                                                                  |
| `pnpm docker:build --to @hcengineering/pod-front --force` | 270 пакетов, 0 ошибок                                                                            |
| `POST /guestJoin` без имени                               | `400 {"error":"Please enter your first and last name"}`                                          |
| `POST /guestJoin` без `personToken`                       | `200` + токен, `roomName`, `person`; лог `No personToken from client, derived one from the link` |
| `/guestInfo` по ссылке                                    | `200`, `canStart: true` при включённом флаге                                                     |

Как повторить API-проверку: временный скрипт в `services/love` (уже удалён), который через `@hcengineering/server-token` подписывает гостевой токен ключом
`createHmac('sha256', 'secret').update('love-guest-token').digest('hex')` и постит на `http://localhost:8096/guestJoin`.
Модуль `@hcengineering/server-token` — CommonJS: импортировать как `import pkg from '...'; const { generateToken } = pkg`.
Рабочие данные стенда: workspace `тест` = `096313e3-58dd-46ea-b91b-c14b2e066b49`, встреча `6ae46775578e687fee18994f`.
Таблицы: `docker exec dev-postgres-1 psql -U postgres -d postgres` (схемы `global_account`, `public`), встречи — в `public.space` (`_class = 'love:class:MeetingMinutes'`).

## Регулярные встречи: где остановились (2026-11-05)

Ветка платформы `FUSIO-1307-livekit-shared-link`, правки не закоммичены. Change:
`openspec/changes/love-recurring-meetings` (статусы в `tasks.md`, решения — в `design.md` §Состояние).

### Что сделано кодом (проверено сборкой и unit-тестами)

- Создание регулярной встречи из диалога события: тумблер «Регулярная встреча» + период
  (`MeetingData.svelte`), `PermanentMeeting` с `repeatEvery` и замена разового события на серию
  (`makeRecurring` в `plugins/love-resources/src/utils.ts`).
- Период: единый модуль `plugins/love-resources/src/period.ts` (`periodRules` / `ruleToPeriod` /
  `retimedRules`), 10 юнит-тестов в `src/__tests__/meetingPeriod.test.ts` — зелёные.
- Карточка серии (`EditScheduledMeeting`): live-подписка, сводка правила, редактор периода
  (переписывает `rules` с сохранением `endDate`/`count`), переключатель напоминания.
- Постоянная встреча (`EditPermanentMeeting`): live-подписка, переименование синхронизируется с серией
  через новое поле миксина `permanentMeeting`.
- Бейдж встречи: extension-point `calendar.extensions.EventCardExtensions` + `MeetingEventBadge`;
  регистрация модельным документом и миграция `meeting-event-card-badge` (`models/love/src/migration.ts`).
- Удаление серии завершает живую сессию: `TxRemoveDoc` в `OnEventUpdate` + 3 теста.
- Напоминание за 15 минут: `reminderBefore` в миксине, `meetingReminders.ts` (клиентский тик + Notification).
- Локали: 10 новых ключей во всех 12 `plugins/love-assets/lang/*.json`.

### Где именно остановились

- Последняя правка — вынос периода в `period.ts` и обновление импортов. После неё **прогнан только
  `get_errors` (чисто)**; полный набор `pnpm build --to @hcengineering/love-resources`,
  eslint по изменённым файлам и `npx jest` в `plugins/love-resources` — **не выполнялся**
  (команда была прервана).
- Образ `@hcengineering/workspace-service` собран (в нём модель + миграции), но контейнер
  `workspace` не пересоздан — значит на стенде модель ещё старая.
- E2E не гонялся: стенд sanity (`tests/docker-compose.yaml`, http://localhost:8083, workspace `sanity-ws`)
  не поднят, доступа к дев-аккаунту 8087 у агента нет.
- Не воспроизведён пункт владельца «период меняется не реактивно при создании ссылки для гостей» —
  нужен сценарий (какое поле, какой шаг).

### С чего продолжить завтра

1. Прогнать проверки после `period.ts`:
   `pnpm build --to @hcengineering/love-resources --to @hcengineering/model-love`,
   `npx eslint` по изменённым файлам, `npx jest` в `plugins/love-resources` (ожидаемо 42 теста).
2. Применить модель на стенде:
   `docker compose -f dev/docker-compose.yaml up -d workspace --force-recreate`
   и убедиться по логам, что прошёл апгрейд воркспейса и состояние `meeting-event-card-badge`.
3. Пересобрать фронт и пересоздать контейнер: `pnpm docker:build --to @hcengineering/pod-front` +
   `up -d front --force-recreate`.
4. Ручной прогон сценария: создать регулярную встречу (период 2 недели) → в планировщике видны вхождения →
   в карточке серии меняется период и напоминание → переименование постоянной встречи меняет имя серии →
   удаление серии при живой сессии завершает её.
5. E2E: поднять `tests/docker-compose.yaml` и прогнать
   `tests/sanity/tests/love/meetings.recurring.tests.ts` (+ `calendar-recurring.spec.ts`),
   разбор — `node analyze_failures.js`. Если стенд поднимать не хочется — смоук на 8087
   с дев-аккаунтом (нужны логин/пароль).
6. Остаток по change: `3.1` (wkst в редакторе), `3.4` (политика ссылки в одном месте),
   `4.2`, `4.4`, `4.5`, `5.2–5.4`, `5.5` (регресс-набор).

## Что осталось сделать

### Проверить руками (5 минут, нужен твой браузер)

1. `Ctrl+Shift+R` на http://localhost:8087, открыть гостевую ссылку на встречу с включённым «Начинать по ссылке может любой»: гость входит и **остаётся** в комнате больше минуты (это и был баг «ссылка сразу закрывается»).
2. Зайти владельцем в ту же комнату — оба в одной комнате, встреча `Active`.
3. Под ролью `User` — кнопка «Добавить комнату» есть; под `Maintainer` — есть и «Редактировать офис».
4. Обычный участник (не владелец) вне окна расписания — встреча не подхватывается (это ожидаемо, так задумано).

### Бэклог по этой фиче

- **Гость-участник.** Залогиненный пользователь, открывший гостевую ссылку, входит как гость и получает вторую персону («Зайцев,Андрей 2»). Решение: если есть токен workspace — входить как участник.
- **Тесты.** Нет юнит-кейсов на owner-исключение в `/getToken` и на `findJoinableScheduled`; прогон `pnpm --filter @hcengineering/pod-love run test` за владельцем репозитория (локально не запускался).
- **LiveKit в compose.** Сейчас контейнер руками; стоит добавить сервисом в `dev/docker-compose.yaml` вместе с конфигом.
- **`bundle:template` cpSync.** Периодически падает `EBUSY`/`unlink` на `services/love/bundle/template/index.html`; лечится `rm -rf services/love/bundle/template`, но лучше сделать идемпотентное копирование с ретраем.
- **Документация.** Страница `communication/invite-link` в `platform-docs` помечена `stale` — текст про «ссылку можно открыть только в окне встречи» больше не верен.
- **Мёртвый код.** Удалить или оживить `EditMeetingData.svelte`.
- **`/guestJoin` без `personToken`** — теперь работает через выведенный id; при желании можно вернуть строгий 400 вместо выведенного.

### Процесс

- Коммиты не делаем (правило пользователя). Форматирование (`pnpm run format`) — тоже за пользователем.
- Дальше по плану — блоки этапов 1–8 в `plan.md`, но они относятся к ветке `FUSIO-1307` (модель F1/F4), на этой ветке их применять некуда.

## Грабли, которые уже стоили времени

- `pnpm docker:build` без `--force` отдаёт «Docker built (cached)» — новые правки не попадают в образ.
- Имена пакетов: сервис love — это `@hcengineering/pod-love` (каталог `services/love`), а `@hcengineering/love` — клиентский плагин.
- Svelte 4 не парсит TS-аннотации в разметке: `on:click={(e: MouseEvent) => ...}` даёт `parse-error`; типизировать только функцию в `<script>`.
- Модель изменилась — без пересборки `pod-love` сервис видит устаревший `types/*.d.ts`.
- Фронт: изменения видны только после пересборки образа `pod-front` и пересоздания контейнера (dev-server не используется).
