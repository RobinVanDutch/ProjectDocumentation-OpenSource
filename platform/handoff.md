# Handoff: гостевая ссылка на встречу (FUSIO-1307)

Состояние на 2026-11-02. Продолжать с этого места.

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
