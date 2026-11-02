# Сверка PR #428 со спецификацией

Источник: `intabia-fusion/platform#428` — «FUSIO-1307: Recurrent meetings», автор `haiodo`.
Ветка `origin/FUSIO-1307`, коммит `46537cd2c8`.
Разбор: 2026-11-02, по 108 файлам патча (+5486 / −913).

Сверяется с: `plan.md` (этапы 0–8), `openspec/changes/love-meetings-followups/` (proposal, spec, tasks),
`openspec/changes/love-meeting-link/` (гостевая ссылка на ветке `FUSIO-1307-livekit-shared-link`).

Статусы в таблицах: **реализовано** — есть в PR и покрыто тестом; **частично** — код есть, но
поведение уже требований или не доказано; **нет** — не сделано; **иначе** — сделано в другой форме;
**новое** — есть в PR, нет в спеке.

---

## 1. Что фактически делает PR

| Область                | Объём        | Суть                                                                                                                                |
| ---------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| `plugins/**` (71 файл) | +2422 / −323 | `love`, `love-resources`, `calendar`, `calendar-resources`, 12+12 локалей                                                           |
| `services/love` (13)   | +1496 / −146 | `resolveSession`, `findMeetingTarget`, гостевые потоки, пароль, rate limit, `getTxOperations`                                       |
| `tests/sanity` (13)    | +646 / −346  | новые `meetings.recurring`, `meetings.permanent`, `calendar-rsvp`; удалены `meetings.scheduled-links`, `meetings.scheduled-connect` |
| `models/**` (4)        | +546 / −3    | `TMeeting`/`TMeetingMinutes`/`TPermanentMeeting`, три миграции                                                                      |
| `server-plugins` (3)   | +151 / −60   | `OnEventUpdate` love, `rsvpSummaryTxes` календаря                                                                                   |
| `docs` (2)             | +204 / −19   | `docs/love.md` §7a, `docs/memory/love-meeting-links.md`                                                                             |

Ключевые решения PR, которых не было в спеке или которые её меняют:

1. **Статуса `MeetingStatus.Scheduled` больше нет.** Сессия не существует, пока её не создаст `resolveSession`.
   `finishMeeting` терминален, re-arm убран.
2. **Идентичность встречи — миксин `MeetingEventLink` на мастер-событии.** `room`/`meetingId` в миксине объявлены
   legacy и «ещё пишутся». Сессия ссылается на серию через `eventId`, на постоянную встречу — через `meeting`.
3. **Постоянная встреча — отдельный класс `PermanentMeeting extends Space`** (F4): без расписания, открыта всегда,
   `occurrence` у её сессии нет.
4. **Ссылка — указатель, а не пропуск.** Payload: JSON `{ eventId, workspace, linkVersion, kind? }`
   (`buildMeetingLinkPayload`/`parseMeetingLinkPayload`), старый JWT читается как legacy.
   Отзыв — бамп `linkVersion`, `checkMeetingLink` → `revoked`/`expired`.
5. **Политика доступа `MeetingAccess`**: `start: 'members' | 'link'`, `past: 'none' | 'last'`,
   `afterTtl` (по умолчанию 7 дней, `null` — бессрочно), `guestPassword: { hash, salt }`.
6. **Пароль гостя — серверный**: `POST /meetingPassword`, pbkdf2-sha256 100 000 итераций, `timingSafeEqual`,
   rate limit 10 промахов / 15 минут на «ссылка + IP» → `429`.
7. **HTTP-контракты с авторизацией**: `GET /meetingLink`, `POST /meetingPassword`, `POST /resolveSession`
   проходят общий гейт `authorizeTarget` (401 без токена, 403 не участнику, 404 нет встречи).
8. **RSVP календаря** (вне объёма нашей спеки): `Event.rsvp` в копии участника, `Event.rsvpSummary` на мастере,
   сводку собирает серверный триггер (`rsvpSummaryTxes`, `collectRsvp`).
9. **Служебный этаж** `love.ids.ScheduledFloor` + `love.ids.ScheduledRoom` (`RoomType.Scheduled`) вместо служебной
   комнаты на клиенте.
10. **Навигатор — URL-сегмент** `/love/<segment>`, `segment ∈ meetings | permanent | <floorId>`, переход сбрасывает `fragment`.

---

## 2. Сверка с этапами `plan.md`

| Этап                           | Ожидание в плане                                                                          | Факт PR #428                                                                                                                                                                                                                                                                            | Статус                                                                        | Что осталось                                                                              |
| ------------------------------ | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| 0. Стенд и базовая линия       | Отчёт базового прогона, список известных падений                                          | В PR нет; прогоны не приложены                                                                                                                                                                                                                                                          | **нет**                                                                       | Собрать стенд на базе ветки PR, прогнать unit + love/calendar sanity, зафиксировать флаки |
| 1. Отмена и перенос вхождения  | `isCancelled` вычитать, время брать из `originalStartTime`                                | `meetingOccurrences` = rules + `rdate` − `exdate` (`plugins/love/src/utils.ts:4787`); override **не учитывается** — в коде прямо: «Overrides are not subtracted: an override is a session of its own» (`:4786`). Unit на `exdate` есть, e2e отмены/переноса нет (`exdate: []` в sanity) | **частично**                                                                  | Перенос: связать override с окном вхождения; e2e на `exdate` и перенос                    |
| 2. Миксин только на мастере    | Отсекать `access !== owner` и `ReccuringInstance`, не считать миксин на инстансе митингом | Миграция `strip-meeting-mixin-from-non-masters` (`models/love/src/migration.ts:986`), ранний `continue` в `OnEventUpdate` (`server-plugins/love-resources/src/index.ts:5228`), разрешение мастера через `meetingMasterQuery` (`plugins/love/src/utils.ts:4743`)                         | **реализовано** (запись не отклоняется, а игнорируется — спеке это разрешено) | Тест «миксин на инстансе не создаёт вторую ссылку» (сейчас только миграционный)           |
| 3. Права постоянной встречи    | Владельца нельзя убрать из `members`, проверка на сервере                                 | Валидации `members` у `PermanentMeeting` нет; `createSession` берёт `members` из `participants`, `owners` = автор события; старый код «добавить создателя в members» удалён                                                                                                             | **нет**                                                                       | Серверная проверка + UI                                                                   |
| 4. Навигатор: сегмент и кнопки | Не резать путь при открытии документа; одна кнопка настроек                               | Введён `OfficeNavigator` с URL-сегментом (`loc.path[3]`, сброс `fragment`), `Hall` рендерит разделы; `navigateToOfficeDoc`/`backOrOpen` **не менялись** — обрезка до `path.length = 3` осталась; `MeetingMinutesSection` получил `query`/`label`                                        | **частично**                                                                  | 4.1 не сделано; 4.2 проверить дубль настроек; `virtual-office.mdx` не в PR                |
| 5. Внеплановый вход по ссылке  | Вне окна вхождения участник (и гость при `link`) может начать встречу                     | Вне окна `resolveSession` возвращает `no-occurrence` + `nextOccurrence` (`services/love/src/workspaceClient.ts:7290`); `start` влияет только на право гостя **создать сессию внутри окна** (`openOnLink` в `guests.ts`). Всегда открыта только `PermanentMeeting` — другой класс        | **нет**                                                                       | Решение и реализация; см. §4                                                              |
| 6. Язык и гостевые режимы      | Язык из интерфейса; гость постоянной встречи без `after`                                  | `CreatePermanentMeetingPopup` пишет `language: 'en'`; `/guestInfo` отдаёт `mode = Finished ? 'after' : …` без исключения для `kind === 'meeting'`                                                                                                                                       | **нет** (1 из 2 пунктов)                                                      | Дефолт языка; запрет `after` для постоянной встречи                                       |
| 7. Чистка мёртвых полей        | Удалить `linkId` и `meetingId`                                                            | `linkId` **заново добавлен** и задокументирован, но нигде не пишется: `models/love/src/index.ts:228,382`, `plugins/love/src/types.ts:4620,4689`; `TMeetingSchedule { room, meetingId }` на месте (`models/love/src/index.ts:400`)                                                       | **нет, объём вырос**                                                          | Удалить `linkId`, либо начать его писать; убрать `TMeetingSchedule.meetingId`/`room`      |
| 8. Сквозная проверка           | Отчёт: 0 реальных падений, статус по этапам                                               | Есть таблица тестов в `docs/love.md` (44/214/28/27/32/9), но прогона и отчёта нет                                                                                                                                                                                                       | **нет**                                                                       | Прогон и отчёт                                                                            |

**Вывод по плану:** PR #428 закрывает этап 2 целиком и часть этапа 1 (отмена), вводит новую механику
ссылок/доступа/постоянных встреч, но этапы 3, 5, 6, 7 и 8 остаются; этап 4 закрыт частично, причём
именно тот подпункт, который был главной жалобой (сброс сегмента при открытии протокола), не тронут.

---

## 3. Сверка с требованиями `specs/love-meetings/spec.md`

| #   | Requirement                                             | Факт PR                                                                                                                                                                                             | Статус                          |
| --- | ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------- |
| 1   | Отменённое вхождение не открывает сессию                | `exdate` вычитается в `meetingOccurrences`; `resolveOccurrence` вернёт `current` пустым, `next` — следующее действительное                                                                          | **реализовано** (unit; e2e нет) |
| 2   | Перенесённое вхождение открывается в новое время        | Override не учитывается: окно считается по расписанию мастера; `occurrence` сессии = `originalStartTime` только когда сессия создалась в окне старого времени                                       | **нет**                         |
| 3   | Миксин встречи только на мастере                        | Миграция + `continue` в триггере + `meetingMasterQuery`; запись не отклоняется, но вторая встреча не появляется                                                                                     | **реализовано**                 |
| 4   | Владелец постоянной встречи остаётся участником         | Проверки нет ни на сервере, ни в UI                                                                                                                                                                 | **нет**                         |
| 5   | Язык постоянной встречи из интерфейса                   | Хардкод `'en'` в `CreatePermanentMeetingPopup.svelte`                                                                                                                                               | **нет**                         |
| 6   | Гость постоянной встречи видит только `before` и `live` | Без сессии — `before` ✔; после `Finished` — `mode: 'after'` и для постоянной встречи ✘                                                                                                              | **частично**                    |
| 7   | Сессия создаётся на вхождение и не переиспользуется     | `createSession` под `apply.notMatch` + `LIVE_MEETING_STATUSES`, сессия несёт `occurrence`; `finishMeeting` терминален; есть тесты на переиспользование живой и на новую сессию следующего вхождения | **реализовано**                 |
| 8   | Ссылка позволяет начать встречу вне окна вхождения      | `resolveSession` отдаёт `no-occurrence` вне окна, `403 «Meeting is not open»`                                                                                                                       | **нет**                         |
| 9   | Отзыв ссылки не зависит от хранения ссылки              | `checkMeetingLink`: несовпадение `linkVersion` → `revoked`; legacy без версии = 1; `afterTtl` считается только когда будущих вхождений нет                                                          | **реализовано**                 |
| 10  | Модель не содержит мёртвых полей ссылки                 | `linkId` добавлен и не пишется; `meetingId` в `TMeetingSchedule`                                                                                                                                    | **нарушено**                    |

Требования 8 и 10 следовало бы вынести из этой дельты в отдельные изменения: 8 — не готова к реализации
(см. §11), 10 — это чистка модели поверх PR.

---

## 4. Внеплановый вход: как это работает в PR на самом деле

Это самое важное расхождение, потому что этап 5 был в плане как P1.

| Шаг                 | PR #428                                                                                           | Наша спека ожидала                         |
| ------------------- | ------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| Окно                | `[occ.start − 15 мин, occ.start + 4 ч)` (`SCHEDULED_JOIN_LEAD_MS`, `SCHEDULED_MEETING_WINDOW_MS`) | то же                                      |
| Серия, окно открыто | Участник → `resolveSession(create=true)` создаёт сессию                                           | совпадает                                  |
| Серия, окна нет     | `no-occurrence` + `nextOccurrence` → `403 { error: 'Meeting is not open', nextOccurrence }`       | Старт разрешён при политике `start`        |
| Гость, окно открыто | `resolveSession(create = start === 'link')`; при `members` — `not-started` → лобби `before`       | совпадает                                  |
| Постоянная встреча  | Открыта всегда: `occurrence = undefined`, сессия создаётся по любому входу                        | «постоянная переговорка» = `start: 'link'` |

То есть `meetingAccess.start` в PR — **право гостя открыть сессию внутри окна**, а не право начать встречу
вне окна. Постоянной переговоркой в PR служит `PermanentMeeting`, а не серия с `start: 'link'`.
Формулировку `docs/love.md` («`link` делает митинг постоянной комнатой») в этой части надо читать как
«гость может открыть сессию сам», иначе поведение расходится с кодом.

---

## 5. Что PR сделал иначе, чем предполагала спека

1. **`MeetingStatus.Scheduled` удалён.** В плане и в `love-meeting-link` фигурирует статус `Scheduled` и re-arm
   после завершения (`restartMeeting`) — этого механизма больше нет: сессия создаётся входом, а не заранее.
2. **`startOnLink` (ветка `FUSIO-1307-livekit-shared-link`) заменён на `meetingAccess.start`.** Поле `startOnLink`
   в модели PR отсутствует. Если PR вольётся, change `love-meeting-link` надо закрыть как поглощённый.
3. **«Одна живая сессия на серию» — неверно.** PR явно допускает две одновременные живые сессии серии
   (затянувшееся вхождение + старт следующего), мастер досыпает участников во все живые, override — только в свою.
   Инвариант держится как «одна живая сессия на вхождение / на постоянную встречу».
4. **`meetingScheduledDate` → `occurrence` + `eventId`.** Миграции: `meeting-settings-to-mixin`,
   `strip-meeting-mixin-from-non-masters`, `create-scheduled-floor`; старые `Scheduled`-сессии удаляются.
5. **RSVP реализован** — открытый вопрос плана («на серию или на вхождение») закрыт: ответ пишется в копию
   участника (у virtual-вхождения — через `updateReccuringInstance`), сводку считает сервер на мастере.
6. **Гостевой лобби другой:** `passwordRequired`, `mode: before | live | after`, `roomFound`, `title`, `now`.
   Кнопки «Начать сейчас» в PR нет — при `start: 'link'` гость просто создаёт сессию входом.
7. **Служебный этаж/комната — объекты модели** (`ScheduledFloor`, `ScheduledRoom`), а не клиентский трюк;
   это новая сущность для документации и для UI-тестов.

---

## 6. Новое в PR, чего нет в спеке (кандидаты в отдельные capability)

| Возможность                     | Где                                                                         | Что описать в спеке                                                                                      |
| ------------------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Пароль гостя                    | `services/love/src/passwords.ts`, `POST /meetingPassword`, `GuestJoinPopup` | хранение (pbkdf2 100k, соль), снятие через `null`, `401`/`429`, что видит гость до ввода пароля (ничего) |
| Rate limit гостевого входа      | `services/love/src/guests.ts` (`10` попыток / `15 мин`, ключ «ссылка + IP») | окно, ключ, ответ `429`, сброс после успеха                                                              |
| RSVP                            | `plugins/calendar*`, `server-plugins/calendar-resources`                    | кто отвечает и где хранится, как считается сводка, `pending`                                             |
| Служебный этаж                  | `models/love` (`create-scheduled-floor`), `isServiceFloor`/`isServiceRoom`  | что скрыто из навигатора и почему                                                                        |
| Авторизация HTTP API            | `authorizeTarget` в `main.ts`                                               | `401/403/404`, кто вправе выдать ссылку и поставить пароль                                               |
| `PermanentMeeting` как документ | `models/love` (Activity, ObjectEditor, вьюлеты)                             | панель встречи, прошедшие сессии во футере, создание из «Зала»                                           |

---

## 7. Дефекты и риски самого PR

Проверено по патчу; каждый — кандидат в задачу.

| #   | Место                                                         | Проблема                                                                                                                                                                                                                                                                 |
| --- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | `plugins/calendar-resources/src/components/EventRsvp.svelte`  | `invited = participants.filter((it) => it !== me)` — `participants: Ref<Contact>`, `me` — документ сотрудника; сравнение по идентичности не сработает, организатор попадёт в `invited`, `pending` завышен на 1. Серверный `collectRsvp` сравнивает корректно (по `user`) |
| 2   | `MeetingLinkPopup`                                            | после `setGuestPassword` в состояние пишется фиктивный `{ hash: '', salt: '' }` — «пароль задан» держится на пустом хеше (`access = { ...access, guestPassword: { hash: '', salt: '' } }`)                                                                               |
| 3   | `plugins/love-resources/src/utils.ts` (`getEventMeetingLink`) | любые ошибки глушатся в `''` (`console.error` + `return ''`); вызывающие не отличают «ссылки нет» от «сервис упал» (`createMeeting` проверяет `link !== ''`, попап — нет)                                                                                                |
| 4   | `CreatePermanentMeetingPopup`                                 | `language: 'en'`, `private: true` захардкожены; роли не проверяются                                                                                                                                                                                                      |
| 5   | `OfficeNavigator`                                             | `canAddFloor = hasAccountRole(...)` вычисляется через `const` при создании компонента — не реактивно; `Main.svelte` кастует сегмент URL в `Ref<Floor>` без проверки                                                                                                      |
| 6   | `linkId`                                                      | добавлен в модель и описан в `docs/love.md` («shortId постоянной ссылки»), но не пишется нигде — документация врёт                                                                                                                                                       |
| 7   | `guests.ts` (`passwordAttempts`)                              | `Map` не ограничен по размеру (в коде помечено `ponytail:`) — рост на уникальных ссылках/IP                                                                                                                                                                              |
| 8   | `docs/love.md`                                                | ссылки на `images/love-permanent-create.png` и `images/love-meeting-link.png`, которых в PR нет                                                                                                                                                                          |
| 9   | Тесты                                                         | нет негативных проверок: «ссылку выдаёт только участник» (`isMeetingParticipant`), «`/meetingPassword` только участник», отзыв legacy-JWT, истечение 15-минутного окна rate limit                                                                                        |
| 10  | Sanity                                                        | серия создаётся с `exdate: []` — отмена и перенос вхождения не проверяются e2e; «владелец в `members`» не проверяется нигде                                                                                                                                              |
| 11  | `server-plugins/love-resources` `OnEventUpdate`               | в тесте фейк возвращает одну живую сессию — веер на несколько сессий и фильтр по `occurrence` для override не покрыты                                                                                                                                                    |

Проверено и снято как ложное: `copyLink` передаёт `getEventMeetingLink(...)` (Promise) в `copyTextToClipboard` —
эта функция принимает `string | Promise<string>` (`packages/presentation/src/utils.ts:883`), так что копирование работает.

---

## 8. Бэклог на основе PR

Порядок — по цене ошибки, не по объёму.

**P1 — ломает штатный сценарий**

1. **Перенесённое вхождение** (этап 1). `meetingOccurrences` не видит override: перенесённая встреча открывается
   в старое время. Нужно решение, откуда брать новые времена (override-документы `ReccuringInstance` против
   `rdate`): сервисный аккаунт должен уметь их читать из `PersonSpace` участников.
   Точки: `plugins/love/src/utils.ts` (`meetingOccurrences`, `resolveOccurrence`), `services/love/src/workspaceClient.ts` (`findMeetingTarget`).
2. **Владелец постоянной встречи** (этап 3). Серверная проверка `members` у `PermanentMeeting` + блокировка в UI.
3. **Язык постоянной встречи** (этап 6). Дефолт из интерфейса, `language` в `CreatePermanentMeetingPopup`.
4. **E2E на отмену и перенос** — сейчас единственное доказательство это unit `resolveOccurrence`.

**P2 — доводка поведения и модель**

5. **Навигатор: путь.** `navigateToOfficeDoc` и `backOrOpen` не должны резать `loc.path` до 3 — иначе протокол
   во время звонка сбрасывает выбранный пункт (главная жалоба из отчёта).
6. **`linkId`**: либо писать при выдаче ссылки (`createMeetingLink`/`revoke`), либо удалить из модели,
   типов и `docs/love.md`. Сейчас — документированное, но неработающее поле.
7. **Мёртвые поля модели**: `TMeetingSchedule.meetingId`, `room` в миксине (legacy) — решение и чистка.
8. **Дефекты UI из §7**: `copyLink` без `await`, фиктивный `guestPassword`, `EventRsvp.pending`,
   реактивность `canAddFloor`.

**P3 — документация и тесты**

9. `virtual-office.mdx` (en, ru): навигатор, служебный этаж, «Forever», ссылка постоянной встречи, скриншоты.
10. `calendar-rsvp.md` — на него уже ссылается `docs/love.md`, файла нет.
11. Тесты из §7 п.9–11 (негативные проверки API, веер сессий, отмена/перенос в sanity).

**Отдельно:** этап 5 (внеплановый старт) из P1 перевести в «решение не принято» — PR показал, что механика
другая (`no-occurrence` вне окна, постоянная переговорка = `PermanentMeeting`). Прежде чем реализовывать,
нужно решить, нужен ли старт вне окна вообще.

---

## 9. Что делать со спецификацией

1. `openspec/changes/love-meetings-followups/tasks.md` — проставить статусы: этап 2 выполнен PR,
   этап 1 частично (только `exdate`), этапы 3/5/6/7/8 остаются, этап 4 частично.
2. `openspec/changes/love-meeting-link/` — пометить как поглощённое PR #428: `startOnLink` и re-arm
   в `Scheduled` в модели PR отсутствуют; если PR вливается, ветка `FUSIO-1307-livekit-shared-link` закрывается.
3. Завести новый change под остаток (**предложение**: `love-meetings-pr428-gaps`) с дельтами:
   перенос вхождения, владелец постоянной встречи, язык, навигатор (путь), `linkId`/мёртвые поля.
   RSVP, пароль и rate limit описать там же отдельными capability (`calendar-rsvp`, `guest-access`),
   потому что в спеке их нет вообще.
4. После приёмки — `openspec archive love-meetings-followups` (дельты уедут в `openspec/specs/`).
5. Этап 5 из `plan.md` убрать из обязательных или переписать под фактическую семантику `meetingAccess.start`
   (§4), иначе спека будет требовать поведения, которого нет и которое не согласовано.

---

## 10. Как проверять

```bash
# база
pnpm build --to @hcengineering/love
pnpm test --to @hcengineering/love          # plugins/love, models/love
cd services/love && pnpm test               # 214 тестов по docs/love.md §11
cd server-plugins/love-resources && pnpm test
cd plugins/love-resources && pnpm test

# стенд и e2e
cd tests/sanity
pnpm run uitest tests/love/meetings.recurring.tests.ts
pnpm run uitest tests/love/meetings.permanent.tests.ts
pnpm run uitest tests/calendar/calendar-rsvp.spec.ts
node analyze_failures.js
```

Отдельно руками (не покрыто тестами): навигатор «в звонке → протокол → назад», панель свойств миксина
(`meetingAccess`, приватность, язык, запись, транскрипция), гость постоянной встречи после завершения сессии.

---

## 11. Открытые вопросы

1. **Внеплановый старт (этап 5).** Нужен ли он вообще, если постоянная переговорка уже есть как
   `PermanentMeeting`? Если нужен — менять `resolveSession` и переписывать требование 8 в спеке.
2. **Перенос вхождения.** Читать override-документы из `PersonSpace` участников (сервисный аккаунт) или
   договориться, что перенос пишет `rdate`/`exdate` в мастер? От этого зависит и объём, и права.
3. **`linkId`.** Писать (тогда зачем, если shortId и так стабилен по payload) или удалять.
4. **`PermanentMeeting`: `after` для гостя.** Запрещать (как в спеке) или оставить — тогда правится спека.
5. **База для остатка работ.** Текущая ветка `FUSIO-1307-livekit-shared-link` не содержит коммита PR
   (`git merge-base --is-ancestor 46537cd2c8 HEAD` → нет). Реализацию остатка логично вести от `FUSIO-1307`.

---

## 12. Что применено на ветке `FUSIO-1307-livekit-shared-link` (2026-11-02)

Вопрос 5 из §11 решён: PR накатан на текущую ветку, а не наоборот.

**Способ.** `git cherry-pick -n 46537cd2c8` — без коммита, изменения остались в индексе и рабочем дереве
(коммит делает владелец репозитория). `pnpm install --frozen-lockfile` — lock от PR слился сам.

**Конфликты (22 файла).** Наша ветка реализовала отменённую PR-ом механику (`startOnLink`,
`MeetingStatus.Scheduled`, re-arm завершённой встречи), поэтому конфликты — это одни и те же места:

| Файлы | Решение |
|---|---|
| `plugins/love/src/{types,plugin}.ts`, `love-resources/src/meetings.ts`, `guest/GuestMeetingApp.svelte`, `EditMeetingData.svelte`, `Floor.svelte`, `FloorConfigure.svelte`, `services/love/guests.ts`, `services/love/__tests__/guests.test.ts`, `docs/love.md` | версия PR |
| `plugins/love-assets/lang/*.json` (12) | сведены обе стороны (ключи develop + ключи PR), `StartOnLink` удалён |
| `plugins/love-resources/src/components/EditMeetingMinutes.svelte` | сохранён guard `handleLeaveRoom` из develop, убраны `connectLabel` и тумблер `startOnLink` |

**Доделано после автослияния** — то, что git слил, но что больше не компилируется или теряет фичу develop:

| Место | Правка |
|---|---|
| `services/love/src/main.ts` | из `/getToken` убраны ветки `MeetingStatus.Scheduled` и вызов `restartMeeting` |
| `services/love/src/workspaceClient.ts` | удалён мёртвый `restartMeeting` |
| `models/love/src/index.ts` | удалено поле `startOnLink` (в типах его уже нет — PR) |
| `plugins/love/src/plugin.ts` | возвращён id `MeetingPrivateEditor` (develop), который PR выкинул; иначе модель ссылается на несуществующий компонент |
| `plugins/love-resources/src/components/guest/GuestMeetingApp.svelte` | возвращён guard `leaving` из FUSIO-1584 (PR его не знал) |
| `tests/sanity/package.json` | убран дубль зависимости `@hcengineering/calendar` |
| `tests/sanity/tests/love/meetings.rage-click.tests.ts` | setup переписан на миксин-модель (событие + `MeetingEventLink`, сессию создаёт сервис); тест «reset» удалён — кнопка исчезла вместе с `Scheduled` |

**Наши практики поверх PR** (пункты §8, которые не требуют нового решения):

| Что | Где | Пункт спеки |
|---|---|---|
| Гость без имени → `400`; без клиентского `personToken` — стабильный `guest-link:<sha256(встреча:имя)>` (+ 3 теста) | `services/love/src/guests.ts`, `__tests__/guests.test.ts` | наш фикс, потерянный при слиянии |
| Язык постоянной встречи из интерфейса вместо `'en'` | `CreatePermanentMeetingPopup.svelte` | этап 6.1 |
| Навигатор: `navigateToOfficeDoc` и `backOrOpen` не режут `path` до 3 сегментов — протокол во время звонка не сбрасывает выбор | `plugins/love-resources/src/utils.ts`, `RoomPopup.svelte` | этап 4.1 |
| RSVP: `pending` по id персоны, организатор не попадает в «приглашённые» | `calendar-resources/src/components/EventRsvp.svelte` | дефект PR §7.1 |
| Попап ссылки: вместо фиктивного `guestPassword: {hash:'',salt:''}` — явный флаг `passwordSet` | `MeetingLinkPopup.svelte` | дефект PR §7.2 |

**Проверки после применения.** Сборка: `@hcengineering/love`, `model-love`, `love-resources`,
`calendar-resources`, `pod-love`, `server-love-resources` — 0 ошибок. Тесты: `services/love` 223,
`plugins/love` 45, `models/love` 9, `server-plugins/love-resources` 34, `plugins/love-resources` 32,
`plugins/calendar` 80, `plugins/calendar-resources` 7 — зелёные. prettier + eslint по изменённым файлам чисто,
конфликтных маркеров нет.

**Что остаётся решить (по-прежнему открыто):** перенос вхождения (§11.2), «владелец всегда участник»
(нужен выбор: ответная транзакция из триггера или валидация), `linkId` (§11.3), гость и `after`
у постоянной встречи (§11.4), мелочи §7.3, §7.4, §7.8.
