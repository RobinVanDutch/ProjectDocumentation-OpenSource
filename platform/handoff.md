# Handoff: гостевая ссылка на встречу (FUSIO-1307)

Состояние на 2026-11-05. Продолжать с этого места.

## 2026-11-05: остаток по регулярным созвонам заведён отдельным изменением

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
